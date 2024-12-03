# PostgreSQL BigQuery Foreign Data Wrapper

This repository demonstrates how to set up and use a Foreign Data Wrapper (FDW) in PostgreSQL to query Google BigQuery tables directly. This setup allows you to seamlessly integrate BigQuery data into your PostgreSQL database.

## Prerequisites

- PostgreSQL 12 or higher
- `postgres_fdw` extension
- Google Cloud Service Account with BigQuery access
- BigQuery dataset and table you want to query

## Installation

1. Enable the required PostgreSQL extension:
```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
```

2. Create a server connection to BigQuery:
```sql
CREATE SERVER bigquery_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (
        host 'www.googleapis.com',
        port '443',
        use_remote_estimate 'true'
    );
```

3. Set up user mapping with your BigQuery credentials:
```sql
CREATE USER MAPPING FOR CURRENT_USER
    SERVER bigquery_server
    OPTIONS (
        "service_account" '{
            "type": "service_account",
            "project_id": "your-project-id",
            "private_key_id": "your-key-id",
            "private_key": "your-private-key",
            "client_email": "your-service-account@project.iam.gserviceaccount.com",
            "client_id": "your-client-id"
        }'
    );
```

## Usage

### Creating a Foreign Table

```sql
CREATE FOREIGN TABLE bigquery_sales (
    id INTEGER,
    customer_name VARCHAR(100),
    sale_date DATE,
    amount NUMERIC(10,2)
)
SERVER bigquery_server
OPTIONS (
    project 'your-project-id',
    dataset 'your_dataset',
    table 'sales_table'
);
```

### Querying Data

```sql
-- Basic query
SELECT * FROM bigquery_sales WHERE sale_date >= '2024-01-01';

-- Join with local tables
SELECT 
    bs.customer_name,
    bs.amount,
    local_customer.region
FROM bigquery_sales bs
JOIN local_customer ON bs.customer_name = local_customer.name;
```

## Optimization Techniques

### Materialized Views

Create materialized views for frequently accessed data:

```sql
CREATE MATERIALIZED VIEW sales_monthly AS
SELECT 
    date_trunc('month', sale_date) as month,
    sum(amount) as total_sales
FROM bigquery_sales
GROUP BY 1;

-- Refresh when needed
REFRESH MATERIALIZED VIEW sales_monthly;
```

### Table Options

Optimize foreign table performance with additional options:

```sql
CREATE FOREIGN TABLE bigquery_sales (
    -- columns as before
)
SERVER bigquery_server
OPTIONS (
    project 'your-project-id',
    dataset 'your_dataset',
    table 'sales_table',
    fetch_size '1000',
    use_remote_estimate 'true',
    updatable 'false'
);
```

## Data Type Mapping

Example of mapping BigQuery data types to PostgreSQL:

```sql
CREATE FOREIGN TABLE bigquery_complex (
    id INTEGER,
    json_data JSONB,  -- BigQuery STRUCT
    array_data TEXT[], -- BigQuery ARRAY
    timestamp_col TIMESTAMP, -- BigQuery TIMESTAMP
    decimal_col NUMERIC(20,4), -- BigQuery NUMERIC
    geography_col TEXT -- BigQuery GEOGRAPHY
)
SERVER bigquery_server
OPTIONS (...);
```

## Best Practices

1. **Security**
   - Keep service account credentials secure
   - Use least privilege access for the service account
   - Regularly rotate credentials

2. **Performance**
   - Monitor query costs (BigQuery charges by data scanned)
   - Implement appropriate partitioning strategies
   - Use materialized views for frequently accessed data
   - Test different fetch sizes for optimization

3. **Maintenance**
   - Regular monitoring of connection status
   - Implementing error handling
   - Keeping track of BigQuery usage and costs

## Error Handling

Example function for safe querying:

```sql
CREATE OR REPLACE FUNCTION query_bigquery_safe(query_text TEXT)
RETURNS TABLE (LIKE bigquery_sales)
AS $$
BEGIN
    RETURN QUERY EXECUTE query_text;
EXCEPTION
    WHEN fdw_error THEN
        RAISE NOTICE 'BigQuery FDW error: %', SQLERRM;
        RETURN;
END;
$$ LANGUAGE plpgsql;
```

## Troubleshooting

Common issues and solutions:

1. Connection failures
   - Verify service account credentials
   - Check network connectivity
   - Ensure proper permissions in BigQuery

2. Performance issues
   - Review query optimization
   - Check fetch size settings
   - Monitor BigQuery quota usage