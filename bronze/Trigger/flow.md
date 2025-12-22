flowchart TD
    %% =========================
    %% Pipeline Start & Logging
    %% =========================
    A[Pipeline Triggered] --> B[Start Master Pipeline Log]

    %% =========================
    %% Metadata Lookups
    %% =========================
    B --> C[Lookup Module Name]
    C --> D[Lookup Dependency Level]

    %% =========================
    %% Dependency-Level Loop
    %% =========================
    D --> E[For Each Dependency Level]

    %% =========================
    %% Dependency Condition
    %% =========================
    E --> F{Dependency Condition Met?}

    %% =========================
    %% TRUE Path - Execute Child
    %% =========================
    F -->|Yes| G[Invoke Child Ingest Pipeline]
    G --> H[Capture Child Execution Status]

    %% =========================
    %% FALSE Path - Skip
    %% =========================
    F -->|No| I[Skip Execution for This Dependency]

    %% =========================
    %% Merge Paths
    %% =========================
    H --> J[Evaluate Error Flag]
    I --> J

    %% =========================
    %% Error Handling Branch
    %% =========================
    J -->|Error Detected| K[Set Error Variable]
    K --> L[Generic Error Handling Script]
    L --> M[Update Master Pipeline Fail Log]
    M --> N[Fail Pipeline]

    %% =========================
    %% Success Path
    %% =========================
    J -->|No Error| O[Continue to Next Dependency Level]

    %% =========================
    %% Loop Completion
    %% =========================
    O --> P{More Dependency Levels?}
    P -->|Yes| E
    P -->|No| Q[End Master Pipeline Log]

    %% =========================
    %% Pipeline End
    %% =========================
    Q --> R[Pipeline Completed Successfully]

