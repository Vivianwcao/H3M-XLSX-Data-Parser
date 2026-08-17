# H3M Field Invoice Parser

A serverless Python microservice on AWS Lambda that parses unstructured `.xlsx` field invoices in memory, converts visual grid headers and line items into clean JSON payloads, and relays data to downstream systems.

---

## Tech Stack

* **Runtime & Compute:** Python 3.13, AWS Lambda, AWS SAM, Amazon API Gateway
* **Data Processing:** NumPy, Pandas (`AWSSDKPandas-Python313` Layer), `openpyxl`, `boto3`
* **Storage & Execution:** Amazon S3 (`emi-v3` bucket), Asynchronous Lambda Invocation
---

## Pipeline Architecture

```mermaid
%%{init: {'themeVariables': { 'edgeLabelBackground': '#F8FAFC' }}}%%
flowchart TD
    %% Node Definitions with 2-line titles for spacious bubbles
    A("API Gateway Webhook<br/>POST /h3m_parser")
    B("AWS Lambda Service<br/>H3M_xlsx_parser")
    C[("Amazon S3 Bucket<br/>emi-v3 Storage")]
    
    subgraph In-Memory ["In-Memory Parsing Engine"]
        D("Pandas & NumPy Engine<br/>io.BytesIO Buffer")
        E("Header Parser<br/>Extract Ticket Metadata")
        F("Line Item Parser<br/>Extract Service Line Items")
    end

    G("Downstream Output Service<br/>PHP Processing Lambda")

    %% Data Flow Connections with 2-line labels
    A --> B
    B -->|"1. Stream Raw .xlsx<br/>File Object"| C
    B -->|"2. Load Bytes to<br/>Memory Buffer"| D
    D -->|"3. Locate Variable<br/>Header Cells"| E
    D -->|"4. Parse Line<br/>Item Rows"| F
    B -->|"5. Save Formatted<br/>JSON Output"| C
    B -->|"6. Trigger Asynchronous<br/>Event Relay"| G

    %% Blueprint Color Classes
    classDef trigger fill:#E0F2FE,stroke:none,color:#0369A1,rx:14px,ry:14px;
    classDef compute fill:#E2F1E6,stroke:none,color:#14532D,rx:14px,ry:14px;
    classDef storage fill:#FEF9C3,stroke:#EAB308,stroke-width:2px,color:#713F12;
    classDef process fill:#DBEAFE,stroke:none,color:#1E40AF,rx:14px,ry:14px;
    classDef relay fill:#FFEDD5,stroke:none,color:#7C2D12,rx:14px,ry:14px;

    %% Category Assignments
    class A trigger;

    %% AWS Lambda Compute
    class B compute;

    %% Amazon S3 Storage Cylinder
    class C storage;

    %% Pastel Blue internal process bubbles (No borders)
    class D,E,F process;

    %% Downstream Relay Service
    class G relay;

    %% Subgraph Container: Borderless, Soft Slate Fill
    style In-Memory fill:#F1F5F9,stroke:none,rx:18px,ry:18px,color:#334155;
```
---

## Parsing & Processing Logic

The service ingests inbound API Gateway webhooks and processes vendor Excel billing sheets through four core execution steps:

* **In-Memory File Streaming:** Fetches target `.xlsx` objects directly from S3 into Python `io.BytesIO` buffers, avoiding local disk write operations inside Lambda.
* **Dynamic Line Item Table Mapping:** Scans rows sequentially using set operations to identify table headers (`Line Item Description`, `Name`, `Qty`, `Unit`, `Rate`, `Subtotal`). The engine dynamically registers column indices and extracts line items regardless of vertical offset until encountering a total row marker.
* **Asynchronous Relay & Output Storage:** Writes the formatted JSON output back to Amazon S3 for auditing and triggers a downstream PHP processing Lambda asynchronously without blocking the client HTTP response.
