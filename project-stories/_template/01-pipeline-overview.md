# Pipeline Overview

One diagram showing the whole pipeline, start to end, so a reader gets the overall shape before any layer file goes into detail. Keep it to main building blocks only — depth belongs in the numbered layer files that follow.

```mermaid
flowchart LR
    subgraph Sources
        A[Source system]
    end

    subgraph Ingestion
        B[Ingestion tool]
    end

    subgraph Storage
        C[(Raw / lake storage)]
    end

    subgraph Transformation
        D[Transform tool]
    end

    subgraph Serving
        E[Warehouse or mart]
        F[BI tool]
    end

    A --> B --> C --> D --> E --> F
```

One or two sentences under the diagram, plain language, naming what each block is in this project.
