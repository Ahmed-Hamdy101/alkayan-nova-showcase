# Alkayan Nova (PHP Project)

This repository contains a document for Alkayan Construction Nova, it runs on business tier, built with a simple MVC-like structure (controllers, models, views) and a public entry point under `public/`.

## Features

- Project pages grouped by type (delivered / planned / under construction / interior / supplies)
- JSON API endpoint for projects (used by frontend rendering)
- Server-side routing via a single front controller (`public/index.php`)
- Basic security hardening headers

## Project Structure

- `public/` — Web root / front controller and public assets
  - `public/index.php` — main router/dispatcher + early JSON API handling
- `app/` — application code
  - `controllers/` — request handlers
  - `models/` — database queries and domain logic
  - `views/` — page templates and view components
  - `helper/` — shared helpers (e.g., assets)
- `config/` — configuration and database connection
- `errors/` — static error pages
- `cache/` — runtime cache files
- `storage/` — storage uploads

## Local Development

1. Ensure PHP + MySQL are available.
2. Update database credentials in `config/config.php`.
3. Point your web server to this project root (see `/.htaccess` and `public/.htaccess`).
4. Visit:
   - `/<type>` pages (e.g., `/?type=construction&view=list` depending on routing usage)

## JSON API (Projects)

The application exposes a projects API directly from `public/index.php`.

Example:

- `/?type=api&resource=projects&tid=1003&page=1`

This returns JSON for the requested project type id (`tid`) and pagination.

## System Architecture

```mermaid
graph TB
    Client["Client Browser"]
    
    subgraph WebServer["Web Server (Apache/Nginx)"]
        FrontController[".htaccess Router"]
    end
    
    subgraph PublicDir["public/"]
        IndexPHP["index.php<br/>(Front Controller)"]
        Assets["Static Assets<br/>(CSS, JS, IMG)"]
    end
    
    subgraph AppLayer["app/"]
        Controllers["Controllers/"]
        Models["Models/"]
        Views["Views/"]
        Helpers["Helpers/"]
    end
    
    subgraph ConfigLayer["config/"]
        ConfigPHP["config.php"]
        DatabasePHP["Database.php"]
        CachePHP["Cache.php"]
    end
    
    subgraph StorageLayer["Storage/"]
        Uploads["uploads/"]
        Cache["cache/"]
        Logs["logs/"]
    end
    
    Database[(MySQL Database)]
    
    Client -->|HTTP Request| FrontController
    FrontController -->|Route| IndexPHP
    IndexPHP -->|Render| Views
    IndexPHP -->|Fetch Data| Controllers
    Controllers -->|Query| Models
    Models -->|Connect| ConfigLayer
    ConfigLayer -->|Query| Database
    Models -->|Return Data| Controllers
    Controllers -->|Pass Data| Views
    Views -->|HTML/JSON| IndexPHP
    IndexPHP -->|Response| Client
    IndexPHP -->|Store| StorageLayer
    
    style Client fill:#e1f5ff
    style PublicDir fill:#fff3e0
    style AppLayer fill:#f3e5f5
    style ConfigLayer fill:#e8f5e9
    style StorageLayer fill:#fce4ec
```

## Entity-Relationship Diagram

```mermaid
erDiagram
    PROJECT_TYPES ||--o{ PROJECTS : "has many"
    PROJECTS ||--o{ DELIVERED_PROJECTS : "inherits"
    PROJECTS ||--o{ PLANNED_PROJECTS : "inherits"
    PROJECTS ||--o{ UNDER_CONSTRUCTION_PROJECTS : "inherits"
    PROJECTS ||--o{ INTERIOR_DESIGN_PROJECTS : "inherits"
    PROJECTS ||--o{ SUPPLIES_PROJECTS : "inherits"
    PROJECTS ||--o{ PROJECT_IMAGES : "contains"
    PROJECTS ||--o{ PROJECT_DETAILS : "has"

    PROJECT_TYPES {
        int tid PK "Type ID"
        string name "Project Type Name"
        string slug "URL Slug"
        text description "Description"
        timestamp created_at
        timestamp updated_at
    }

    PROJECTS {
        int id PK "Project ID"
        int tid FK "Project Type ID"
        string title "Project Title"
        string slug "URL Slug"
        text description "Project Description"
        string location "Project Location"
        date start_date "Start Date"
        date end_date "End Date"
        string status "Status"
        int display_order "Display Order"
        timestamp created_at
        timestamp updated_at
    }

    DELIVERED_PROJECTS {
        int id PK "Project ID"
        string client_name "Client Name"
        float budget "Project Budget"
        int completion_percentage "Completion %"
    }

    PLANNED_PROJECTS {
        int id PK "Project ID"
        date planned_start "Planned Start"
        date planned_end "Planned End"
        string priority "Priority Level"
    }

    UNDER_CONSTRUCTION_PROJECTS {
        int id PK "Project ID"
        int progress_percentage "Progress %"
        date estimated_completion "Est. Completion"
        string current_phase "Current Phase"
    }

    INTERIOR_DESIGN_PROJECTS {
        int id PK "Project ID"
        string style "Design Style"
        string designer_name "Designer Name"
        string materials "Materials Used"
    }

    SUPPLIES_PROJECTS {
        int id PK "Project ID"
        string supplier "Supplier Name"
        int quantity "Quantity"
        string category "Supply Category"
    }

    PROJECT_IMAGES {
        int id PK "Image ID"
        int project_id FK "Project ID"
        string image_path "Image Path"
        string alt_text "Alt Text"
        int display_order "Display Order"
    }

    PROJECT_DETAILS {
        int id PK "Detail ID"
        int project_id FK "Project ID"
        string key "Detail Key"
        text value "Detail Value"
    }
```

## Authentication Flow

> **Note:** Authentication and identity management are decoupled from this core application. Authentication services have been migrated to a dedicated subdomain (`realstate.alkayan-co.com`) and are operated as an independent microservice/server module.

## API Endpoints & Request-Response Flow

### Request-Response Flow

```mermaid
sequenceDiagram
    participant Browser as Browser
    participant IndexPHP as public/index.php
    participant Router as Router Logic
    participant ProjectController as ProjectController
    participant ProjectModel as Project Model
    participant DB as Database

    Browser->>IndexPHP: GET /?type=api&resource=projects&tid=1003
    IndexPHP->>Router: Parse Query Parameters
    
    alt type == api
        Router->>ProjectController: Handle API Request
        ProjectController->>ProjectModel: getProjectsByType(tid=1003)
        ProjectModel->>DB: SELECT * FROM projects WHERE tid=1003
        DB-->>ProjectModel: Return Records
        ProjectModel-->>ProjectController: [Project Array]
        ProjectController->>ProjectController: Build JSON Response
        ProjectController-->>IndexPHP: JSON Data
        IndexPHP->>Browser: Response:<br/>{<br/>  "success": true,<br/>  "data": [...],<br/>  "count": N<br/>}
    else Regular Page Request
        Router->>ProjectController: Handle Page Request
        ProjectController->>ProjectModel: Fetch Data
        ProjectModel->>DB: Query Data
        DB-->>ProjectModel: Return Data
        ProjectModel-->>ProjectController: Data
        ProjectController->>ProjectController: Render View
        ProjectController-->>IndexPHP: HTML
        IndexPHP->>Browser: Rendered HTML Page
    end
```

## Data Flow Diagrams

### Project Listing Data Flow

```mermaid
graph TD
    A["User Visits<br/>/?type=construction&view=list"] -->|Request| B["public/index.php<br/>(Front Controller)"]
    
    B -->|Parse Route| C{Route Type?}
    
    C -->|API Request| D["ProjectController<br/>handleAPI()"]
    C -->|Page Request| E["ProjectController<br/>handlePage()"]
    
    D -->|Query| F["Project Model<br/>getProjectsByType()"]
    E -->|Query| F
    
    F -->|SQL Query| G["MySQL Database"]
    G -->|Result Set| F
    
    F -->|Projects Array| D
    F -->|Projects Array| E
    
    D -->|json_encode| H["JSON Response"]
    E -->|Pass to View| I["View Template<br/>construction/list.php"]
    
    I -->|Render HTML| J["HTML Output"]
    
    H -->|Content-Type: JSON| K["Response to Browser"]
    J -->|Content-Type: HTML| K
    
    K -->|Display| L["User Browser"]

    style A fill:#e1f5ff
    style B fill:#fff3e0
    style D fill:#f3e5f5
    style E fill:#f3e5f5
    style F fill:#e8f5e9
    style G fill:#fce4ec
    style L fill:#e1f5ff
```

### File Upload Data Flow

```mermaid
graph TD
    A["User Submits<br/>Upload Form"] -->|POST| B["public/index.php"]
    
    B -->|Check type=upload| C["UploadController<br/>handleUpload()"]
    
    C -->|Validate File| D{File Valid?}
    
    D -->|No| E["Return Error<br/>Message"]
    D -->|Yes| F["Generate Filename<br/>storage/uploads/"]
    
    F -->|Store File| G["File System<br/>storage/uploads/"]
    
    G -->|Success| H["Database Operation"]
    
    H -->|INSERT file_record| I["MySQL Database"]
    
    I -->|Confirm| J["Redirect/Response"]
    
    E -->|Error| J
    
    J -->|Response| K["User Browser"]

    style A fill:#e1f5ff
    style B fill:#fff3e0
    style C fill:#f3e5f5
    style G fill:#fce4ec
    style I fill:#fce4ec
    style K fill:#e1f5ff
```

## Component Interaction

### Component Dependency Map

```mermaid
graph TB
    subgraph External["External Dependencies"]
        PHP["PHP Runtime"]
        MySQL["MySQL Database"]
        WebServer["Web Server<br/>(Apache/Nginx)"]
    end
    
    subgraph Core["Core Components"]
        FrontController["Front Controller<br/>(public/index.php)"]
        Router["Router<br/>(Route Dispatcher)"]
    end
    
    subgraph Business["Business Logic Layer"]
        Controllers["Controllers<br/>(Request Handlers)"]
        Models["Models<br/>(Data Access)"]
        Helpers["Helpers<br/>(Utilities)"]
    end
    
    subgraph Presentation["Presentation Layer"]
        Views["Views<br/>(Templates)"]
        Assets["Static Assets<br/>(CSS/JS/IMG)"]
    end
    
    subgraph Infrastructure["Infrastructure Layer"]
        Config["Configuration<br/>(Database, Cache)"]
        Cache["Cache System<br/>(File-based)"]
        Storage["Storage<br/>(Uploads, Logs)"]
    end
    
    WebServer -->|Route Request| FrontController
    FrontController -->|Parse & Dispatch| Router
    
    Router -->|Route to| Controllers
    Controllers -->|Query Data| Models
    Controllers -->|Utility| Helpers
    
    Models -->|Connect via| Config
    Config -->|Query| MySQL
    
    Controllers -->|Pass Data| Views
    Views -->|Reference| Assets
    
    Controllers -->|Store| Storage
    Models -->|Read/Write| Cache
    
    PHP -->|Executes| FrontController
    PHP -->|Executes| Controllers
    PHP -->|Executes| Models
    
    style External fill:#ffebee
    style Core fill:#fff3e0
    style Business fill:#f3e5f5
    style Presentation fill:#e8f5e9
    style Infrastructure fill:#fce4ec
```

### Controller Interaction Diagram

```mermaid
graph TB
    subgraph Controllers["Available Controllers"]
        CC["ConstructionController"]
        IC["InteriorDesignController"]
        SC["SuppliesController"]
        PC["PlannedProjectController"]
        DPC["DeliveredProjectController"]
        UC["UnderConstructionController"]
        ContactC["ContactController"]
        UploadC["UploadController"]
    end
    
    subgraph SharedResources["Shared Resources"]
        PM["Project Model"]
        Helper["Assets Helper"]
        Config["Database Config"]
    end
    
    CC -->|Query via| PM
    IC -->|Query via| PM
    SC -->|Query via| PM
    PC -->|Query via| PM
    DPC -->|Query via| PM
    UC -->|Query via| PM
    
    CC -->|Use| Helper
    IC -->|Use| Helper
    SC -->|Use| Helper
    PC -->|Use| Helper
    DPC -->|Use| Helper
    UC -->|Use| Helper
    
    PM -->|Connect via| Config
    
    ContactC -->|Store| Config
    UploadC -->|Store Files| Config

    style Controllers fill:#f3e5f5
    style SharedResources fill:#e8f5e9
```

### Request Processing Workflow

```mermaid
graph LR
    A["HTTP Request"] -->|1. Receive| B["Front Controller<br/>(public/index.php)"]
    B -->|2. Parse| C["Extract Parameters<br/>type, view, resource, etc"]
    C -->|3. Route| D{Route Type?}
    
    D -->|Page| E["Load View Controller"]
    D -->|API| F["Load API Handler"]
    D -->|Upload| G["Load Upload Handler"]
    
    E -->|4a. Fetch Data| H["Model Query"]
    F -->|4b. Fetch Data| H
    G -->|4c. Process| I["File Operations"]
    
    H -->|5. Query DB| J["Database"]
    J -->|6. Return Data| H
    
    H -->|7a. Data| E
    H -->|7b. Data| F
    I -->|7c. Result| G
    
    E -->|8. Render| K["View Template"]
    F -->|8. Encode| L["JSON Encode"]
    G -->|8. Redirect| M["Response"]
    
    K -->|9. HTML| M
    L -->|9. JSON| M
    
    M -->|10. Send| N["HTTP Response"]
    N -->|Display| O["User Browser"]

    style A fill:#e1f5ff
    style B fill:#fff3e0
    style O fill:#e1f5ff
```

## License

MIT License. See `LICENSE`


