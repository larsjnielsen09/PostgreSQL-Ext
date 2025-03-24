# PostgreSQL-Ext: Your Real-time, Function-Driven Backend on PostgreSQL

## System Overview and Explanation (For LLMs and Developers)

PostgreSQL-Ext is an open-source framework that transforms PostgreSQL into a real-time, function-driven application backend, similar in concept to Convex.dev, but built entirely on the foundation of PostgreSQL and its extension capabilities. It aims to provide a streamlined development experience while maintaining the flexibility, control, and data integrity benefits of a self-hosted, PostgreSQL-centric architecture.  It avoids vendor lock-in to a specific cloud provider.

**Core Concepts:**

*   **Database as the Core:** PostgreSQL is the central component, serving as the single source of truth for data, schema, and server-side logic.  This contrasts with traditional architectures where a separate application server handles much of the business logic.
*   **Real-time Data Synchronization:**  Changes to the database (inserts, updates, deletes) are automatically and efficiently propagated to connected clients (web browsers, mobile apps) in real-time, without requiring client-side polling.
*   **Function-Driven Backend:**  Server-side logic is implemented as PostgreSQL functions, primarily using PL/pgSQL.  These functions are stored *within* the database and executed directly by the database server, providing transactional guarantees and close proximity to the data.
*   **Automatic API Generation:**  A RESTful API is automatically generated from the database schema using PostgREST, eliminating the need to write boilerplate CRUD (Create, Read, Update, Delete) code.  GraphQL support is optionally available via the `pg_graphql` extension.
*   **Simplified Development Workflow:** A command-line interface (CLI) tool, `pgext`, streamlines common development tasks, such as project initialization, schema migrations, function deployment, and local development environment setup.

**Architectural Components:**

1.  **PostgreSQL Database (Extended):**
    *   Standard PostgreSQL (version 14 or later recommended) with its core features: tables, indexes, views, roles, row-level security (RLS), etc.
    *   **`pg_events` (Custom Extension):**  This is the *key* component that enables real-time functionality.  It's a PostgreSQL extension (written in C) that provides:
        *   **Change Data Capture (CDC):**  Efficiently detects changes to specified tables *without* relying solely on triggers. It likely leverages PostgreSQL's logical replication infrastructure internally.
        *   **Event Queue:**  A lightweight, in-database queue (likely using advisory locks or a dedicated table) to store change events reliably.
        *   **Event Emission:**  Publishes events to a separate websocket server (`pg_events_ws`).
    *   **`pg_graphql` (Optional Extension):** Provides a GraphQL API directly from the database schema.

2.  **`pg_events_ws` (Websocket Server):**
    *   A separate process (likely written in Node.js, Go, or Rust) that maintains persistent websocket connections with clients.
    *   Subscribes to the event queue managed by the `pg_events` extension.
    *   Receives change events from `pg_events`.
    *   Filters events based on client subscriptions and permissions (using information from JWTs).
    *   Broadcasts relevant events to connected clients.
    *   Designed for horizontal scalability (multiple instances behind a load balancer).

3.  **PostgREST (API Layer):**
    *   A standalone web server that automatically generates a RESTful API from a PostgreSQL database schema.
    *   Handles authentication and authorization using PostgreSQL roles and RLS, typically via JWTs (JSON Web Tokens).
    *   Translates HTTP requests into SQL queries and executes them against the database.
    *   Returns results in JSON format.

4.  **`pgext` (CLI Tool):**
    *   A command-line interface for managing PostgreSQL-Ext projects.
    *   Provides commands for:
        *   Project initialization (`pgext init`).
        *   Schema migrations (`pgext migrate`).
        *   Function deployment (`pgext deploy-functions`).
        *   Local development environment setup (`pgext dev`).
        *   Deployment to production environments (`pgext deploy`).
        *   Log viewing (`pgext logs`).

5.  **Frontend Application:**
    *   Standard web application (HTML, CSS, JavaScript) that interacts with the PostgreSQL-Ext backend via:
        *   The PostgREST API (for CRUD operations and calling PL/pgSQL functions).
        *   The `pg_events_ws` websocket server (for real-time updates).

**Data Flow (Example: Adding a Task):**

1.  **Client Request:** The frontend sends a request to the PostgREST API to add a new task (e.g., `POST /tasks`).
2.  **PostgREST:**
    *   Authenticates the request (using JWTs).
    *   Authorizes the request (using PostgreSQL roles and RLS).
    *   Translates the request into a SQL `INSERT` statement.
3.  **PostgreSQL:**
    *   Executes the `INSERT` statement within a transaction.
    *   The `pg_events` extension detects the change.
    *   `pg_events` adds an event to its internal queue.
4.  **`pg_events_ws`:**
    *   Receives the event from the `pg_events` queue.
    *   Determines which connected clients are subscribed to changes to the `tasks` table (and potentially filters based on user ID or other criteria).
    *   Sends the event data (the new task) to the relevant clients via websockets.
5.  **Client Update:** The frontend receives the event and updates the UI to display the new task.

**Key Advantages Over Traditional Architectures:**

*   **Reduced Latency:** Server-side logic (PL/pgSQL functions) runs directly within the database, minimizing network overhead.
*   **Data Consistency:** PostgreSQL's transactional guarantees ensure data integrity.
*   **Simplified Development:**  The `pgext` CLI and automatic API generation reduce boilerplate code.
*   **Real-time Capabilities:** Built-in real-time updates without the complexity of managing separate messaging systems.
*   **Security:** Leverages PostgreSQL's robust security features (roles, RLS).
*   **Open Source and Extensible:** Avoids vendor lock-in and allows for customization.

**This detailed explanation should provide a clear understanding of PostgreSQL-Ext's architecture and functionality for both LLMs and developers.**

---

## Prerequisites

*   **PostgreSQL (14+ recommended):** You'll need a running PostgreSQL instance. This can be:
    *   A local installation.
    *   A managed database service (e.g., AWS RDS, Google Cloud SQL, Azure Database for PostgreSQL).
    *   A Docker container.
*   **Node.js (16+ recommended):** Required for the `pgext` CLI tool and potentially for the websocket server (if you choose Node.js).
*   **Docker (Optional, but recommended):** For local development and simplified deployment.
*   **C Compiler (Only if building `pg_events` from source):**  The `pg_events` extension is written in C.

## Installation

1.  **Install the `pgext` CLI Tool:**

    ```bash
    npm install -g pgext  # (Assuming it's published on npm)
    ```

2.  **Install the `pg_events` PostgreSQL Extension:**

    *   **Pre-built Packages (Recommended):**
        *   If pre-built packages are available for your system (e.g., `.deb` for Debian/Ubuntu, `.rpm` for Red Hat/CentOS), install them using your package manager.  This is the easiest option.  The project would need to provide these.
        *   Example (Debian/Ubuntu): `sudo apt install postgresql-ext-pg-events`
    *   **Building from Source:**
        *   Clone the PostgreSQL-Ext repository:
            ```bash
            git clone https://github.com/your-org/postgresql-ext  # Replace with the actual repository URL
            cd postgresql-ext/packages/pg_events
            ```
        *   Compile the extension (requires a C compiler and PostgreSQL development headers):
            ```bash
            make
            sudo make install
            ```
        *   Enable the extension in your PostgreSQL database:
            ```sql
            CREATE EXTENSION pg_events;
            ```

3.  **Install PostgREST:**

    *   Download the pre-built binary for your system from the [PostgREST releases page](https://github.com/PostgREST/postgrest/releases).
    *   Place the executable in a directory in your `PATH` (e.g., `/usr/local/bin`).
    *   Alternatively, you can install it using a package manager if available (e.g., `apt install postgrest`).

4. **Install Websocket Server (pg_events_ws):**
    * Clone the repository:
        ```
        git clone <repository url for pg_events_ws>
        cd pg_events_ws
        ```
    * Install dependencies and build (example for Node.js):
        ```
        npm install
        npm run build # If a build step is required
        ```
    *   Configure the server (see "Configuration" below).

## Getting Started (Quick Start)

1.  **Initialize a New Project:**

    ```bash
    pgext init my-app
    cd my-app
    ```

    This creates a new project directory with the following structure:

    ```
    my-app/
    ├── schema.sql       (Initial database schema)
    ├── functions/       (Directory for PL/pgSQL functions)
    ├── frontend/        (Directory for your frontend code)
    ├── postgrest.conf  (PostgREST configuration)
    ├── .env             (Environment variables - DO NOT COMMIT)
    └── docker-compose.yml (Optional, for local development)
    ```

2.  **Configure Environment Variables (`.env`):**

    Create a `.env` file in your project directory and add the following:

    ```
    DATABASE_URL="postgresql://user:password@host:port/database_name"
    POSTGREST_JWT_SECRET="your-jwt-secret"  # Generate a strong secret
    WEBSOCKET_SERVER_URL="ws://localhost:3001" # URL of your websocket server
    ```
    *   **`DATABASE_URL`:**  Your PostgreSQL connection string.
    *   **`POSTGREST_JWT_SECRET`:**  A secret key used for JWT authentication with PostgREST.  Generate a strong, random string (e.g., using `openssl rand -base64 32`).
    *   **`WEBSOCKET_SERVER_URL`:** The URL where your websocket server will be running.

3.  **Define Your Schema (`schema.sql`):**

    Edit the `schema.sql` file to define your database tables, columns, relationships, and RLS policies.  See the example in the "Example Application" section below.

4.  **Write PL/pgSQL Functions (`functions/`):**

    Create `.sql` files in the `functions/` directory to define your server-side logic.

5.  **Start the Development Environment:**

    ```bash
    pgext dev
    ```

    This command (provided by the `pgext` CLI) will:

    *   Start a local PostgreSQL database (if you're using Docker Compose).
    *   Apply your schema migrations (using a tool like Flyway, integrated into `pgext`).
    *   Start PostgREST.
    *   Start the `pg_events_ws` websocket server.
    *   Optionally, start a development server for your frontend code.

6.  **Develop Your Frontend:**

    Write your frontend code (HTML, CSS, JavaScript) in the `frontend/` directory.  Use a JavaScript library to connect to the websocket server and the PostgREST API.

7.  **Deploy:**

    ```bash
    pgext deploy
    ```
    This command (provided by the pgext CLI) will handle deploying your schema, functions, and frontend code to your production environment. The specifics of deployment will depend on your chosen infrastructure (see "Deployment" section).

## Configuration

*   **`schema.sql`:**  Defines your database schema using standard PostgreSQL DDL (Data Definition Language).
*   **`functions/`:** Contains your PL/pgSQL functions.
*   **`postgrest.conf`:**  Configures PostgREST.  See the [PostgREST documentation](https://postgrest.org/en/stable/configuration.html) for details.  Key settings include:
    *   `db-uri`:  Your PostgreSQL connection string.
    *   `db-schema`:  The schema to expose via the API (usually `public`).
    *   `jwt-secret`:  Your JWT secret.
    *   `db-anon-role`:  The role to use for unauthenticated requests.
    *   `server-port`: The port PostgREST listens on.
*   **`pg_events_ws` Configuration:**  The websocket server will likely have its own configuration file (e.g., `config.json` or `config.yaml`).  Key settings include:
    *   PostgreSQL connection details.
    *   Port to listen on.
    *   Authentication settings (e.g., JWT secret).
    *   Table/channel subscription rules (optional).

## Example Application (Simple To-Do List)

**`schema.sql`:**

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    title TEXT NOT NULL,
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

CREATE POLICY tasks_user_policy ON tasks
    FOR ALL
    TO authenticated
    USING (user_id = current_setting('request.jwt.claims')::jsonb->>'sub');

CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_tasks_modtime
    BEFORE UPDATE ON tasks
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();


functions/complete_task.sql:

CREATE OR REPLACE FUNCTION complete_task(task_id UUID)
RETURNS tasks AS $$
DECLARE
  updated_task tasks;
BEGIN
    UPDATE tasks
    SET completed = TRUE
    WHERE id = task_id
    RETURNING * INTO updated_task;

    RETURN updated_task;
END;
$$ LANGUAGE plpgsql;
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
SQL
IGNORE_WHEN_COPYING_END

frontend/app.js (Simplified):

// ... (See full example in previous response for details) ...

const postgrestUrl = '/'; // Or your PostgREST endpoint
const websocketUrl = 'ws://localhost:3001'; // Your websocket server URL
const authToken = localStorage.getItem('authToken');

const socket = new WebSocket(websocketUrl);

socket.onopen = () => {
    socket.send(JSON.stringify({ type: 'subscribe', table: 'tasks' }));
    fetchTasks(); // Fetch initial tasks
};

socket.onmessage = (event) => {
    const message = JSON.parse(event.data);
    // ... (Handle insert, update, delete events) ...
};

async function fetchTasks() { /* ... */ }
async function addTask() { /* ... */ }
async function completeTask(taskId) { /* ... */ }
async function deleteTask(taskId) { /* ... */ }
function renderTask(task) { /* ... */ }
function removeTask(taskId) { /* ... */ }
function updateTask(task) { /* ... */ }
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
JavaScript
IGNORE_WHEN_COPYING_END
Deployment

Okay, I've reformatted that section of the README.md to improve readability and clarity. I've used a combination of headings, bullet points, and consistent indentation to make the structure more apparent. I've also made some minor wording improvements.

## Deployment

PostgreSQL-Ext is designed for flexible deployment.  Here are some common options:

### Deployment Options

*   **Docker Compose (Local Development):**
    *   The `pgext dev` command, along with the provided `docker-compose.yml` file, simplifies running a local development environment.
    *   This environment includes PostgreSQL, PostgREST, and the websocket server, all running in Docker containers.

*   **Cloud Providers (AWS, Google Cloud, Azure, etc.):**
    *   Use a managed PostgreSQL service (e.g., AWS RDS, Google Cloud SQL, Azure Database for PostgreSQL).
    *   Deploy PostgREST and the websocket server (`pg_events_ws`) to:
        *   Virtual machines (e.g., EC2, Compute Engine, Virtual Machines).
        *   Container services (e.g., ECS, GKE, AKS).
    *   Use a load balancer to distribute traffic across multiple instances of PostgREST and the websocket server.

*   **Self-Hosted (Bare Metal or VMs):**
    *   Install PostgreSQL, PostgREST, and the websocket server directly on your servers.
    *   Configure `systemd` (or a similar service manager) to manage the processes (start, stop, restart, auto-start on boot).
    *   Set up a reverse proxy (e.g., Nginx, Apache) to handle:
        *   Routing incoming requests to the appropriate services (PostgREST, websocket server).
        *   SSL termination (handling HTTPS).

*   **Platform-as-a-Service (Fly.io, Heroku, etc.):**
    *   Deploy using Docker containers.
    *   Leverage the platform's built-in features for scaling, networking, and database management.

### Deployment Steps (General)

1.  **Set up Infrastructure:** Provision servers, databases, and networking resources as needed for your chosen deployment option.

2.  **Configure Environment Variables:** Set the necessary environment variables on your servers:
    *   `DATABASE_URL`: PostgreSQL connection string.
    *   `POSTGREST_JWT_SECRET`: Secret key for JWT authentication.
    *   `WEBSOCKET_SERVER_URL`: URL of the websocket server.
    *   ... (any other environment variables required by your application or components).

3.  **Deploy PostgreSQL:**
    *   **Managed Service:** Create the database instance using the cloud provider's interface or CLI.
    *   **Self-Hosted:** Install and configure PostgreSQL on your server.

4.  **Deploy `pg_events`:** Install the `pg_events` extension on your PostgreSQL database.

5.  **Deploy PostgREST:** Deploy the PostgREST binary to your server and configure it (using `postgrest.conf`) to connect to your database.

6.  **Deploy `pg_events_ws`:** Deploy the websocket server to your server and configure it to connect to your database and listen for events from `pg_events`.

7.  **Deploy Frontend:** Deploy your frontend code:
    *   **Web Server (Nginx, Apache):** Copy your frontend files to the appropriate directory (e.g., `/var/www/html`).
    *   **Static Site Hosting (Netlify, Vercel):** Follow the provider's instructions for deploying your site.

8.  **Configure DNS and Routing:**
    *   Set up DNS records to point your domain name to your servers.
    *   Configure routing (e.g., using a reverse proxy) to direct traffic to PostgREST, the websocket server, and your frontend.

## CLI Reference (`pgext`)

The `pgext` CLI tool provides the following commands:

*   `pgext init <project-name>`: Initializes a new PostgreSQL-Ext project, creating the directory structure and basic configuration files.
*   `pgext dev`: Starts a local development environment (database, PostgREST, websocket server).
*   `pgext migrate`: Applies schema migrations to the database (using a tool like Flyway, integrated into `pgext`).
*   `pgext deploy-functions`: Deploys PL/pgSQL functions (from the `functions/` directory) to the database.
*   `pgext deploy`: Deploys the entire application (schema, functions, and frontend code) to your production environment.
*   `pgext logs`: Displays logs from the various components (database, PostgREST, websocket server).
*   `pgext db:create`: Creates the PostgreSQL database.
*   `pgext db:drop`: Drops the PostgreSQL database.
*   `pgext db:reset`: Drops, recreates, and migrates the PostgreSQL database.

## Contributing

We welcome contributions! Please see the `CONTRIBUTING.md` file for guidelines.

## License

This project is licensed under the MIT License - see the `LICENSE` file for details.


Key changes and why they improve readability:

Headings: Used ## and ### headings to clearly separate sections (Deployment Options, Deployment Steps, CLI Reference).

Bullet Points: Used bullet points extensively to break down lists of options and steps. This makes it much easier to scan and understand the different choices.

Consistent Indentation: Used consistent indentation to show the hierarchical relationship between items (e.g., sub-bullets under deployment options).

Bold Text: Used bold text for key terms and commands (e.g., pgext dev, DATABASE_URL).

Code Formatting: Used backticks () to format code snippets and filenames (e.g.,docker-compose.yml,schema.sql`).

Explanatory Text: Added short explanations for each item, rather than just listing them. For example, explaining what systemd is used for.

Sub-bullets for Details: Used sub-bullets to provide more detail under main bullet points (e.g., listing different cloud providers).

Separation of Concerns: Separated the deployment options from the steps, to make it clearer.

CLI Reference Formatting: Improved the formatting of the CLI reference section.

This revised formatting should be significantly easier to read and understand, both for humans and for LLMs parsing the document. It presents the information in a more structured and organized way.

